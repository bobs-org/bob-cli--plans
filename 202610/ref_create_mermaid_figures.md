---
tier: epic
title: Render Mermaid diagrams as vector figures in bob ref create PDFs
goal: 'Explicit Mermaid code blocks in Markdown and vault-note imports print as fitted,
  selectable, Highlights-safe vector figures instead of wrapped source. Rendering
  is offline and pinned, costs about one browser launch per document, and a broken
  diagram or missing renderer degrades visibly without ever failing the import.

  '
phases:
- id: adapter
  title: Mermaid `diagrams` op in the web-clip adapter
  depends_on: []
  size: medium
  description: 'adapter: vendor a pinned Mermaid 12.x bundle and a static DejaVu Sans
    TTF, add a `diagrams` op that renders batches offline in one Chrome page to fitted
    vector PDFs with per-item error isolation, report the version from ping, and extend
    the self-test.

    '
- id: ast
  title: Diagram collect and swap filters, wired behind an off-by-default hook
  depends_on: []
  size: medium
  description: 'ast: add the Rust diagrams module with the kill switch, the fence
    pre-scan, the pandoc collect pass, and the results writer, plus the collect and
    swap Lua filters, inserted first in both renderers through RenderFilters but unused
    in production yet.

    '
- id: wire
  title: Render through the adapter on both routes
  depends_on:
  - adapter
  - ast
  size: medium
  description: 'wire: add the adapter client call, orchestration that never fails
    the import, the results and fallback notes, the scale warnings, and the summary
    and warning output on the generic and snapshot routes, with stub-adapter tests
    and an athena smoke run.

    '
- id: finish
  title: Dry-run, doctor, docs, live test, and acceptance
  depends_on:
  - wire
  size: medium
  description: 'finish: add the browser-free dry-run count, the warn-only doctor row,
    the docs, a gated live end-to-end PDF test, the athena acceptance run on the 12-fence
    report, Mac acceptance per decision, the optional decision record, and follow-up
    notes.

    '
decisions:
  mac_check:
    ask: How should the final phase cover the Mac acceptance run?
    choices:
      manual: Phase hands Bryan a Mac checklist; no agent touches the Mac
      ssh: Phase also tries the CLI recreate on the Mac over Tailscale SSH (best-effort)
    default: manual
    why: The Mac is often offline and the Highlights selection check needs Bryan anyway
    answer: manual
  decision_record:
    ask: Add a decisions-web record for the Mermaid rendering and failure policy?
    memory:
    - decisions:ref-create-renders-mermaid-in-the-adapter
    default: false
    answer: false
proposed_by: bbugyi200.athena.research.4r.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 19:29:16
status: done
bead_id: bob-cli-8y
---

- **PROMPT:** [prompts/202610/ref_create_mermaid_figures.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_create_mermaid_figures.md)
- **BEAD:** [bob-cli-8y](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8y/README.md)

# Plan: Mermaid figures in `bob ref create` PDFs

## Context

`bob ref create` renders Markdown through pandoc → XeLaTeX on two routes:

- **Generic Markdown route**: `render_temp_pdf` in
  `src/native/highlights_ref/create.rs`. It runs one pandoc call with pandoc's inferred
  Markdown reader and the `code_break` and `return_links` Lua filters.
- **Vault-note snapshot route**: `render_note_pdf` in
  `src/native/highlights_ref/note_snapshot/profile.rs`. It reads
  `-f commonmark_x-tex_math_dollars-emoji`, adds `note.lua`, writes `.tex` with
  `cwd = scratch`, and then runs XeLaTeX separately for up to three passes.

Neither route knows about Mermaid, so a ` ```mermaid ` fence prints as a wrapped
`fvextra` code block. The screenshot shows a `flowchart LR` with literal `<br/>` tags.
The main caller is unattended: the SASE `research-highlights` file hook runs
`bob ref create --include-id -P …` on every research commit. Research reports hold
nearly all of the demand (dozens of fences); the vault has none today.

The design follows the consolidated research report
`research:202610/ref_create_mermaid_figures/ref_create_mermaid_figures.md`, and Bryan
accepted all of its requirements (A1–A10). In short:

- Render explicit Mermaid code blocks to fitted vector PDFs with a **vendored, pinned
  Mermaid 12.x bundle**.
- Run the bundle inside Bob's **existing Playwright/Chrome web-clip adapter**: one
  offline page and one browser launch per document.
- Swap the PDFs into the **unchanged pandoc → XeLaTeX pipeline** with a first-position
  Lua filter that both routes share.
- When a diagram fails, keep its source under a visible italic note. Never fail the
  import.

Rejected (do not reintroduce): host `mmdc`, the `mmdr` Rust renderer, PNG or SVG print
inputs, printing the whole document through Chromium, hosted renderers (Kroki,
mermaid.ink), aborting on failure, and a new CLI flag.

The research spike's proven snippets live in the research sidecar under
`202610/ref_create_mermaid_figures/ref_create_mermaid_figures__final_assets/`:
`spike_render_mermaid.py`, `spike_collect_mermaid.lua`, and `spike_swap_mermaid.lua`.
Open them with `sase repo open research` (they are not indexed artifacts). Read the
report itself with `sase artifact read`.

## Requirements every phase honors

- **A1 Scope.** Only explicit Mermaid code blocks, as pandoc parses them:
  ` ```mermaid `, `~~~mermaid`, and ` ``` {.mermaid} `, including fences nested in lists
  and quotes. Never sniff `graph` or `flowchart` inside untagged fences. A ` ```text `
  fence remains the way to show source on purpose.
- **A2 Both Markdown routes, one code path.** This covers the generic route and
  vault-note snapshots, including `-M/--plain`. PDF, PDF-URL, arXiv, and web-article
  imports do not change.
- **A3 Degrade per block; never fail, never silent.** A failed block keeps its source,
  gets an italic note ("Mermaid diagram not rendered: Parse error on line 4 …"), and
  produces one warning. If the renderer is unavailable (no `uv`, no browser, adapter
  crash or timeout, collect failure), Bob prints one warning and every block gets the
  note. The exit status is unchanged.
- **A4 Vector, Highlights-safe text.** Use Chrome `page.pdf` with a **static TTF**
  loaded from a `data:` URL; never the adapter's variable woff2 fonts, which embed as
  Type 3. Disable CSS filters so nothing is rasterized.
- **A5 Pinned, local, deterministic.** Use the vendored bundle, a bundled font, and a
  fixed theme. Nothing touches the network during rendering. Extend Mermaid's `secure`
  list with `theme`, `themeCSS`, `themeVariables`, and `fontFamily`, so a diagram's
  `%%{init}%%` directive cannot restyle or unfont the output.
- **A6 No CLI flag.** The only switch is the env kill switch
  `BOB_HIGHLIGHTS_DIAGRAMS=off`, which skips the whole diagram pass: blocks print as
  source with no note.
- **A7 Dry-run stays browser-free; doctor only warns.**
- **A8 Honest layout.** Scale down only. Never rewrite `LR` to `TB`, drop nodes, or
  paginate. Warn when the effective scale drops below 0.5.
- **A9 No backfill.** Existing library PDFs are never re-rendered automatically.
- **A10 Out of scope:** other diagram languages, SVG image embeds, a listen-skip marker,
  a persistent cache, landscape pages, and `accTitle`/`accDescr` alt text.

## Shared contracts (fixed here so phases can run in parallel)

### Adapter op `diagrams` (adapter protocol 1)

Request (Rust → adapter stdin):

```json
{
  "protocol": 1,
  "op": "diagrams",
  "workdir": "/abs/<scratch>/diagrams",
  "items": [{ "id": "<40-hex sha1>", "source": "flowchart LR\n  A --> B" }],
  "item_timeout_s": 10
}
```

- `workdir` is absolute; the adapter creates it (`0700`) and sets `TMPDIR` to it, as
  `op_compose` does. This keeps the Chrome socket path short.
- `id` must match `^[0-9a-f]{40}$`. The adapter writes `<workdir>/<id>.pdf` and nothing
  else.
- At most 200 items (`MAX_DIAGRAM_ITEMS`). Rust never sends more; any extra blocks
  degrade with "too many Mermaid diagrams in one document (limit 200)". The adapter
  rejects more than 200 items as `invalid_request`.

Success response:

```json
{
  "protocol": 1,
  "ok": true,
  "op": "diagrams",
  "mermaid": "12.2.0",
  "browser": { "kind": "chrome", "path": "…", "version": "…" },
  "results": [
    {
      "id": "…",
      "ok": true,
      "pdf": "/abs/…/<id>.pdf",
      "width_pt": 897.0,
      "height_pt": 234.0
    },
    { "id": "…", "ok": false, "error": "Parse error on line 4: …" }
  ],
  "warnings": []
}
```

- A per-item parse error, render error, or timeout is `ok: false` on that item, never an
  op failure.
- Op-level failures use the existing `error_response` shape
  (`{"ok": false, "error": {kind, message, hint}}`). Examples: no browser
  (`kind: "browser"`), launch failure, the font fails to load, or the bundle fails to
  inject.
- `ping` gains `"mermaid": "<bundled version>"`.

### Pandoc-side files under `<scratch>/diagrams/`

- `manifest.json` is written by the collect filter: an array, in document order, of
  `{"id": sha1(text), "text": "…"}`, one entry per Mermaid block occurrence.
- `results.json` is written by Rust and read by the swap filter:
  `{"diagrams": {"<id>": {"pdf": "/abs/path.pdf"} | {"error": "…"}}, "fallback_error": "…" | null}`.
  A block whose id is missing from `diagrams` uses `fallback_error`, or "Mermaid
  renderer unavailable" when that is null.
- The swap filter gets the results path through
  `--metadata bob-diagrams-results=<abs path>`. The collect filter gets the manifest
  path through `--metadata bob-diagrams-manifest=<abs path>`. Both filters use the
  existing `pandoc.json` dependency (`return_links.lua` already requires it).
- Keys are `pandoc.utils.sha1(cb.text)`, computed by pandoc in both passes. Rust never
  hashes, so the keys always agree as long as both passes use the same reader flags.

## Phase `adapter`: Mermaid `diagrams` op in the web-clip adapter

Add the renderer to the pinned Python adapter (`scripts/web_clip/`).

1. **Vendor the bundle.**
   - Add `scripts/web_clip/vendor/mermaid.min.js` (the self-contained IIFE build that
     sets `globalThis.mermaid`). Use the newest 12.x whose fixtures pass; expect 12.2.0.
     The research recorded sha256 `a8a6a6f8…c3b9f`, so verify the prefix and suffix.
     Fetch it with `npm pack mermaid@<ver>` or the registry tarball.
   - Add `MERMAID_LICENSE` (MIT, copied from the package).
   - Add a section to `vendor/README.md` with the version, the source path in the
     package, the full sha256, and the license, using the same pattern as the Defuddle
     entry.
   - Confirm the bundle contains no `import(` calls, so it has no lazy chunks.
2. **Vendor the font.** Add the static **DejaVu Sans Book** TTF as
   `scripts/web_clip/fonts/DejaVuSans.ttf`, plus its license (`DEJAVU_LICENSE.txt`, the
   Bitstream Vera/DejaVu terms). Update `fonts/README.md`. Add Bold only if fixtures
   show that synthesized bold looks wrong.
3. **Embed the assets.** Register every new file in `SUPPORT_ASSETS` in
   `src/scripts.rs`. This includes the new Python module, the bundle, both licenses, and
   the font, with `install_path` under `web_clip/`. Embed them raw; the binary grows by
   about 6 MB, which is accepted.
4. **Sibling module `scripts/web_clip/diagram_render.py`.** Keep it separate so the 118
   KB adapter does not grow, and load it the way `web_clip_render` is loaded (`sys.path`
   insert). It should contain:
   - A browser context with `offline=True`, `service_workers: "block"`, and
     `context.route("**/*", abort)`, so every request is aborted.
   - A page built with `page.set_content(...)` that holds an `@font-face` rule for
     `BobDiagram`, loaded from a `data:font/ttf;base64,…` URL, plus
     `html,body{margin:0;padding:0;background:white}`.
   - The bundle injected with `add_script_tag(content=…)`.
   - An `await document.fonts.load("16px BobDiagram")` call **before** the first render,
     followed by a `document.fonts.check(...)` assertion. If the font did not load, fail
     the op; do not silently fall back to a system font.
   - A `mermaid.initialize` call with: `startOnLoad: false`, `securityLevel: 'strict'`,
     `theme: 'neutral'`, `fontFamily: 'BobDiagram'`, `themeVariables` (with `fontFamily`
     and light note colors replacing neutral's dark-gray notes),
     `themeCSS: '* { filter: none !important; }'`, `suppressErrorRendering: true`, and
     `useMaxWidth: false` for every diagram family the 12.x config schema defines. Also
     pass the default `secure` list extended with `theme`, `themeCSS`, `themeVariables`,
     and `fontFamily`.
   - Per item:
     1. Call `mermaid.render("bob-d<n>", text)` inside a `Promise.race` with
        `item_timeout_s`.
     2. Mount the SVG in an `inline-block` host and measure `getBoundingClientRect`.
     3. Call
        `page.pdf(path, width=f"{w}px", height=f"{h+1}px", print_background=True, page_ranges="1")`
        with zero margins.
     4. Assert that the file is non-empty and starts with `%PDF`.
     5. Report `width_pt = w*0.75` and `height_pt = (h+1)*0.75`.
   - Error strings are the first line of the error, trimmed to 200 characters.
   - After a per-item timeout, close the page and rebuild a fresh one before continuing,
     since the old page may be wedged. A synchronous hang is bounded by the Rust
     transport deadline instead.
5. **`op_diagrams` in `web_clip_adapter.py`.**
   - Validate the request (absolute `workdir`, item shape, id regex, the item cap, a
     positive `item_timeout_s`).
   - Create the workdir and set `TMPDIR`.
   - Find a browser with `discover_browser()`. If there is none, raise
     `AdapterFail("browser", …, BROWSER_HINT)`.
   - Launch with `_launch_with_fallback(..., headless=True)`.
   - Register the op in `OPS` and add it to the module docstring.
   - Add `MERMAID_VERSION` (and `MERMAID_SHA256`) constants next to `DEFUDDLE_VERSION`.
   - Report `mermaid` from `op_ping`.
6. **Self-test and checks.**
   - Extend `--self-test`.
   - Offline: request validation (bad ids, a relative workdir, more than 200 items, an
     unknown field shape).
   - The vendored bundle's sha256 matches `MERMAID_SHA256`.
   - With a discovered browser: the screenshot's flowchart, a sequence diagram, and a
     malformed diagram render in one call. Check that:
     - the good items return one-page PDFs;
     - the bad item is isolated as `ok: false` with "Parse error";
     - no item contains `/Type3` and each one embeds `/FontFile2`;
     - no request reached the network. Use `pdffonts` and `pdfimages -list` when
       available; otherwise byte checks.
   - Without a browser, print `skipped: no browser` and still pass.
   - Add `diagram_render.py` to the `py_compile` list in `just check-web-clip-adapter`.
7. **Pin check.** If 12.2.0 fails any fixture, step back to the newest 12.x that passes.
   Record the reason in `vendor/README.md`.

Done when `just check-web-clip-adapter` passes on athena (browser checks included),
`just check` passes, and a hand-run `diagrams` request through the materialized adapter
renders the screenshot flowchart to a vector PDF whose text `pdftotext` can extract.

## Phase `ast`: diagram collect and swap filters, wired behind an off-by-default hook

Build the pandoc-side half in a new Rust module. Production output must stay
byte-for-byte unchanged until phase `wire` turns the hook on.

1. **New module `src/native/highlights_ref/diagrams.rs`**, registered in `mod.rs`, with
   two embedded Lua files next to it: `diagrams_collect.lua` and `diagrams.lua`. Embed
   them with `include_str!`, as `return_links.lua` is. The module provides:
   - **The kill switch.** `ENV_DIAGRAMS = "BOB_HIGHLIGHTS_DIAGRAMS"`; values `off`, `0`,
     `false`, or `no` (case-insensitive) disable the pass. Read it through
     `crate::native::env`.
   - **Pre-scan.** `prescan(markdown: &str) -> usize` counts candidate fence openers.
     Strip leading whitespace, `>` quote markers, and list markers. A candidate is a run
     of three or more backticks or tildes whose remaining line contains `mermaid`. The
     scan may over-match but must never under-match; documents with no candidates skip
     everything.
   - **Collect pass.** `collect(...)` runs pandoc with **the route's own reader flags**
     (generic: no `-f`, the same path argument, so pandoc infers the reader the same
     way; snapshot: `-f commonmark_x-tex_math_dollars-emoji`). It passes
     `--lua-filter diagrams_collect.lua` and `--metadata bob-diagrams-manifest=…`, and
     writes its output to a throwaway scratch file. It then parses `manifest.json` into
     ordered blocks, de-duplicated by id while counting occurrences.
   - Extract the snapshot reader string into a shared `pub(crate) const` in
     `profile.rs`, so the collect pass and `render_note_pdf` cannot drift.
   - **A results writer** for `results.json` (per the contract), with absolute PDF
     paths.
2. **`diagrams_collect.lua`.** Start from `spike_collect_mermaid.lua`, but read the
   manifest path from metadata, not an env var. Match `cb.classes:includes("mermaid")`
   only.
3. **`diagrams.lua` (the swap filter).** Start from `spike_swap_mermaid.lua`.
   - Read `bob-diagrams-results` from Meta. Wrap the file read and `pandoc.json.decode`
     in `pcall`. If metadata is absent or unreadable, return nothing changed. The swap
     filter must never make pandoc fail.
   - **Rendered block** (entry has a `pdf` that exists) →
     `pandoc.Para{pandoc.Image({}, pdf, "", pandoc.Attr(cb.identifier))}`. Use empty alt
     and no title, so no implicit `Figure N` caption appears. Set no width, so pandoc's
     `\maxwidth`/`\maxheight` (3.1) or `\pandocbounded` (3.2+) scales the image down
     only.
   - **Otherwise** →
     `pandoc.Para{pandoc.Emph(pandoc.Inlines("Mermaid diagram not rendered: " .. reason))}`
     followed by the original `cb`.
4. **Hook in both renderers.**
   - Add `diagrams: Option<DiagramFilter { filter: PathBuf, results: PathBuf }>` to
     `RenderFilters`.
   - When the field is `Some`, `render_temp_pdf` and `render_note_pdf` add
     `--lua-filter <filter>` **before** every other filter, plus
     `--metadata bob-diagrams-results=<abs>`. On the snapshot route, resolve both
     through `resolve_path_for_cwd`.
   - Running first means `note.lua` already centers the resulting `Para{Image}` and
     wraps it in `\BobImageBegin`/`\BobImageEnd`, and `return_links.lua` ignores images.
   - Make `create_markdown_plan` use `write_render_filters` instead of its duplicated
     filter-writing block.
   - Every production caller passes `diagrams: None` in this phase.
5. **Tests.** Gate them the existing way, printing
   `skipping …: pandoc/xelatex is required`.
   - Pre-scan: backtick, tilde, and `{.mermaid}` fences; fences in lists and
     blockquotes; indented fences; a literal example inside a ` ````markdown ` fence
     (counting it is an allowed over-match); ` ```text ` and untagged `flowchart` fences
     are not counted; a document without Mermaid counts 0.
   - Collect: on both reader flag sets, it finds `~~~ {.mermaid}` and list-nested
     fences, ignores untagged ones, and de-duplicates identical sources.
   - Swap, using a fake one-page PDF (build it with `lopdf` or write minimal bytes), on
     both routes through the real `render_temp_pdf` and `render_note_pdf`:
     - the output PDF has no `flowchart LR` text;
     - no `Figure 1` caption appears;
     - the identifier is preserved (`pandoc -t native` check);
     - a missing or error entry produces the italic note followed by the source block;
     - absent or corrupt `results.json` leaves output identical to today;
     - on the snapshot route the image is centered (the `\begin{center}` in the
       generated `.tex`).

Done when `just check` passes and every existing create/snapshot test is unchanged.

## Phase `wire`: render through the adapter on both routes

Turn the pass on: call the adapter, write results, and report.

1. **Adapter client** (`clip_adapter.rs`).
   - Add the `DiagramsRequest`/`DiagramsSuccess`/`DiagramItemResult` serde types (per
     the contract) and `ClipAdapterClient::diagrams(...)`.
   - Add `mermaid: Option<String>` to `PingResponse`.
   - Make `run_request`'s timeout hint parameterizable. Today it hard-codes "check your
     network connection". A diagram timeout should say that a diagram may be too
     complex, that diagrams print as source, and that `BOB_HIGHLIGHTS_DIAGRAMS=off`
     skips rendering.
   - Deadline: `90 s + 10 s × unique items`, capped at `DEFAULT_TIMEOUT_SECS` (300).
     `BOB_WEB_CLIP_TIMEOUT_SECS` overrides it when set.
2. **Orchestration** in `diagrams.rs`:
   `prepare(pandoc, source: DiagramSource, route: DiagramRoute, scratch) -> DiagramPass`.
   It never returns an error.
   - Run the kill switch, then pre-scan, then the collect pass. With the switch on or no
     candidates, return `filter: None` without spawning anything.
   - Resolve the client, then send one `diagrams` call with the unique blocks (the first
     200).
   - Write `results.json`:
     - per-item `pdf` or `error`;
     - `fallback_error` set to the resolve, crash, timeout, or collect failure message
       when the whole renderer is unavailable;
     - over-cap blocks get the limit error.
   - Always return the filter when any Mermaid block was collected, so failures show
     their notes.
   - Use `DiagramSource::Path` (generic: `plan.source`) or `DiagramSource::Text`
     (snapshot: `snapshot.projected_markdown`, written to `diagrams/input.md`).
     `DiagramRoute` carries the reader flags and the text box used for the scale check.
3. **Scale warning (A8).**
   - The effective scale is `min(1, text_width / width_pt, text_height / height_pt)`.
   - Text box per route: the generic route is 489.6 × 669.6 pt (letter, 0.85 in
     margins); the snapshot route is 453.6 × 655.2 pt (1.1 in sides, 0.9/1 in top and
     bottom). Keep these constants next to the geometry arguments they derive from.
   - Below 0.5, warn
     `Mermaid diagram 1 of 12 (flowchart LR) prints at 46% scale; labels may be small`.
4. **Route wiring.**
   - Generic route: run `prepare` in `create_markdown_plan` after the dry-run return and
     the pandoc lookup, before `render_temp_pdf`.
   - Snapshot route: run it in `create_snapshot_plan` before `render_note_pdf`.
   - Pass `filters.diagrams` from the pass. This covers `--listen` (the staged render
     already includes the figures) and `-M/--plain`.
   - Diagram PDFs live under `ScratchDir`, so they outlive every XeLaTeX pass, and
     `BOB_HIGHLIGHTS_KEEP_WORKDIR=1` keeps `diagrams/` with the manifest and results.
5. **Output.** Mirror the return-links pattern.
   - Print one stdout summary line in the success block when blocks exist:
     `diagrams: rendered 11 of 12 Mermaid diagrams`, counting block occurrences.
   - Print warnings to stderr with `styler.warning_prefix()`:
     - one per failed unique diagram
       (`Mermaid diagram 4 of 12 not rendered: Parse error on line 4: …`), or
     - exactly one when the renderer is unavailable
       (`Mermaid diagrams printed as source: <reason>` plus the hint);
     - plus any scale warnings.
   - The exit status never changes because of diagrams.
6. **Tests.** Use stub adapters via `BOB_WEB_CLIP_ADAPTER`. Extend
   `tests/cli/highlights/fake_clip.rs` or use a small stub script that writes bare PDFs
   for the requested ids. Gate on pandoc and xelatex where the full render runs.
   - Success: the summary line appears and the PDF lacks the source text.
   - Partial failure: a note plus source for the bad block, one warning, exit 0.
   - Crash (non-zero exit), invalid JSON, and timeout (a stub that sleeps, with a tiny
     `BOB_WEB_CLIP_TIMEOUT_SECS`): one warning, every block noted, exit 0.
   - Missing `uv`/adapter: one warning, exit 0.
   - Kill switch: the stub is never spawned and the output equals today's.
   - No Mermaid: the stub is never spawned and pandoc runs exactly once.
   - Snapshot route: the same success and failure shape.
7. **Smoke.** With the workspace build (`cargo run` or `target/debug/bob`; never
   `just install`, which replaces Bryan's installed `bob`), hand-run a real import of a
   small Mermaid Markdown file into a scratch vault on athena. Confirm that the figure
   renders and that a broken fence shows a note. Do not import into Bryan's real vault.

Done when `just check` passes and the athena smoke run shows rendered figures plus a
visible note for a broken fence.

## Phase `finish`: dry-run, doctor, docs, live test, and acceptance

1. **Dry-run (A7).** When the pre-scan count is above 0, both `print_plan` (generic) and
   the snapshot dry-run print `diagrams: N Mermaid fence(s) (rendered at create time)`.
   With the kill switch on, they print
   `… (rendering disabled by BOB_HIGHLIGHTS_DIAGRAMS=off)` instead. Dry-run still ends
   with `writes: none` and spawns no pandoc, adapter, or browser. Test this with a
   missing `BOB_PANDOC_COMMAND` and a recording stub adapter that must stay uncalled.
2. **Doctor.** Inside `append_web_clip_doctor_rows`, reuse the same ping; do not add a
   second ping. Add these rows:
   - `mermaid: ok (bundled 12.2.0 · chrome <version>)`;
   - `mermaid: warn (no browser; diagrams print as source)`;
   - `mermaid: warn (adapter ping failed; diagrams print as source)`;
   - `mermaid: off (BOB_HIGHLIGHTS_DIAGRAMS=off)`. Warn rows push onto `warnings` and
     never fail the doctor. Add a test with a stub ping.
3. **Docs** (`docs/highlights-create.md`):
   - a Prerequisites row ("Markdown with Mermaid: `uv` + a browser, optional; without
     them diagrams print as source");
   - a **Diagrams** subsection covering scope (A1), the routes (A2), the failure policy
     with sample output (A3), layout and scale warnings (A8), no backfill (A9), the
     pinned version and how to bump it, and the A10 out-of-scope list;
   - the `--dry-run` table row;
   - a `BOB_HIGHLIGHTS_DIAGRAMS` row in the env-var table;
   - a note in "Scratch staging" about `diagrams/`. Update `README.md` only where it
     lists Markdown prerequisites.
4. **Gated live test.** It skips unless `uv`, a discovered browser, pandoc, xelatex,
   `pdftotext`, `pdffonts`, and `pdfimages` are all present. It runs a real
   `bob ref create` on a fixture Markdown file containing the screenshot flowchart, a
   sequence diagram, a `~~~ {.mermaid}` fence inside a list item, and a malformed fence.
   Assert:
   - `pdftotext` contains "Agent finishes its work" and lacks "flowchart LR";
   - the note text and the malformed source both survive;
   - `pdffonts` shows no Type 3;
   - `pdfimages -list` is empty;
   - exit 0, with the summary line `rendered 3 of 4`. Also cover a source path with
     spaces and non-ASCII characters, and a long-`TMPDIR` regression (a `TMPDIR` over 40
     bytes still renders).
5. **Acceptance on athena.**
   - Open `gh:sase-org/sase--research` with `sase repo open` and import
     `merge_queue_fence_reassessment_explained` (12 fences) into a **scratch** vault
     (`-b`/`-l` pointing at a temp dir; never Bryan's vault, per A9).
   - Check that every diagram renders or fails visibly, the body text is unchanged, and
     the added wall time is about one browser launch (~2–3 s), not per diagram.
   - Record timings and the `pdffonts` and `pdfimages` output in the phase bead notes.
6. **Mac acceptance.**

   > [!decision] mac_check = manual End the phase with a short checklist for Bryan:
   >
   > - recreate the same report on the Mac into a scratch vault;
   > - in Highlights, select and highlight label text inside a diagram;
   > - copy a body-text quote to confirm no Type 3 damage;
   > - confirm a malformed diagram shows its note.

   > [!decision] mac_check = ssh Best-effort: if `ssh mac` connects, build this commit
   > there into a temporary cargo target and run the CLI recreate into a scratch
   > directory. Report `pdffonts` and `pdftotext` results. Bryan still does the
   > Highlights selection checks by hand. If the Mac is offline, record that and fall
   > back to the manual checklist.

7. **Decision record.**

   > [!decision] decision_record Use `/sase_memory_write`, then add a new `decisions`
   > web strand with slug `ref-create-renders-mermaid-in-the-adapter`, following the
   > web's record format (Applies to / Claim / Why / Cost / Reopens when), with
   > `metadata.status: accepted`. It records that explicit Mermaid blocks render through
   > the vendored bundle in the web-clip adapter into vector PDFs, and that failures
   > degrade visibly without failing the import. It names the rejected alternatives
   > (mmdc, mmdr, PNG/SVG, whole-document Chromium, hosted renderers, abort-on-failure)
   > and cites the research report and the landed commits. Then run `sase memory init`.

8. **Follow-ups.** Record each as a `PROPOSED FOLLOW-UP:` note on the phase bead; do not
   create beads:
   - landscape pages for diagrams below 0.5 scale;
   - routing SVG image embeds through the same op;
   - a listen-skip marker for diagrams;
   - a prompt nudge in `sase-research-artifacts` toward `flowchart TD` and short labels;
   - re-evaluating `mmdr` with `svg2pdf` as a browser-free backend behind the same
     `{id, source} → pdf` interface. Also note that the research file hook uses the
     installed `bob`, so the feature reaches it after Bryan's next `just install`. The
     phase does not reinstall.

Done when `just check` and `just check-web-clip-adapter` pass, the live test passes on
athena, and the acceptance results are recorded.
