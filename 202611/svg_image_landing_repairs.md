---
tier: tale
title: Finish SVG image handling and land bob-cli-k6
goal:
  Fix SVG eligibility, ordering, root sanitization, and overwrite diagnostics, then
  verify and close bob-cli-k6.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-k6.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-k6
create_time: 2026-11-11 09:57:24
status: wip
---

- **PARENT:**
  [202611/ref_create_svg_images.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/ref_create_svg_images.md)
- **BEAD:**
  [bob-cli-k6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k6/README.md)

# Finish SVG image handling and land bob-cli-k6

## Scope and completed review

Finish the four bounded defects found while landing **bob-cli-k6**, then complete that
epic's landing in this same coding turn. This tale has no later land agent. Do not wait
for this tale's commit SHA, push, or CI result before closing the epic.

Read `sase bead read bob-cli-k6 -r "Need landing scope and notes"` and the approved plan
with
`sase artifact read plan:202611/ref_create_svg_images.md "Need the approved SVG contract"`.
The epic's plan is `202611/ref_create_svg_images.md` in the plans sidecar. Both phases,
bob-cli-k6.1 and bob-cli-k6.2, are closed with resolution done. The lander reviewed
every phase note, the epic plan, commits `52f0d615` and `7eb82440`, and their source,
Lua filters, adapter, tests, and documentation.

The existing behavior to preserve is vector PDF → same-stem PNG/JPG/JPEG → visible
placeholder with warning. Generic Markdown and `-M` use that pipeline; ordinary
vault-note snapshots keep their current SVG placeholders. Keep protocol 1 and omit
`kind` for Mermaid. Keep one browser launch, per-item failure isolation, the kill
switch, existing Mermaid summary wording, source-directory resolution, and limits. Do
not change the capture hook, host packages, release version, or memory.

The final epic decision is **recapture=yes**. Phase k6.2 note #2 records `just install`
followed by successful `keep_moving_link_marks` recapture (2/2 Mermaid, 1/1 SVG, exit
0). This acceptance was completed; no new production capture is required for these
bounded corrections.

Integration review through fetched `origin/master=7eb82440`: non-epic commits `b2c3697e`
(keep-moving umbrella semantics), `dcc1986e` (ledger display docs), and `ba3f51ba`
(album fixes) do not touch the image pipeline or add a competing SVG renderer.
`ba9867a5`, after epic creation but before its first code commit, adds album reports and
likewise needs no SVG integration. Check any newer drift before closeout; do not undo
those changes.

## 1. Make SVG eligibility conservative and collection ordered

In `src/native/highlights_ref/diagrams.rs`, `prescan_svg` currently returns zero for all
three valid SVG-only Markdown examples below. Actual Pandoc collection returns one SVG
for each, so `prepare` incorrectly skips vector rendering:

```markdown
![Label](<figure name.svg>)
```

```markdown
![Label](figure.svg)
```

```markdown
![Label][fig]

[fig]: figure.svg
```

Use a conservative candidate gate that cannot exclude these Pandoc Image forms; the
Pandoc collector remains authoritative. Prefer a simple permissive SVG marker scan over
a second partial Markdown parser. Keep Mermaid dry-run counts unchanged. Exercise
`prepare` with an adapter stub for these forms, including a real filename with a space,
so tests prove the adapter is reached and source bytes are resolved.

`src/native/highlights_ref/diagrams_collect.lua` uses Pandoc's default typewise
traversal. For a Mermaid fence followed by `![Later](later.svg)`, it emits the SVG
first. Preserve actual document order across both kinds (including nested lists and
quotes), retaining SHA-1 keys, deduplication, per-kind occurrence counts, and the shared
200-item budget. Add alternating Mermaid/SVG cases in both starting orders; the current
test only starts with SVG and therefore misses the defect. Include a budget-boundary
regression showing that later SVGs cannot displace earlier Mermaid items merely because
of their node type.

## 2. Strip event handlers from the SVG root too

In `scripts/web_clip/diagram_render.py::_render_svg_one`, sanitization visits
`root.querySelectorAll('*')`, which excludes the root. An SVG root with
`onmouseover="window.bobRootHandlerRan=true"` retains that attribute at print time; the
lander dispatched a harmless mouseover event immediately before `page.pdf` and observed
the sentinel become true in the existing offline Chrome context. The approved
inline-render fallback requires stripping script elements and all `on*` attributes
before DOM insertion. Apply that rule to the root and descendants. Keep foreignObject
labels selectable and network routes blocked.

Add a browser-backed adapter regression that inspects the inserted root before print and
proves a root event handler cannot execute; retain descendant-handler and script removal
coverage. Verify foreignObject text with the existing live SVG/vector test when its
tools are available. Do not replace the vector renderer with rasterization or weaken the
test gates. For standalone browser probes on Apollo, `TMPDIR=/tmp` avoids the long SASE
temp path exceeding Chromium's Unix socket limit; the first lander probe failed for that
environment reason, while the same probe with the short temp directory succeeded.

## 3. Report SVG results on successful overwrites

In `src/native/highlights_ref/create.rs`, `overwrite_markdown` computes `reports` and
`diagram_pass` but emits neither the image-guard outcome nor the diagram pass. The plain
branch of `overwrite_vault_note` discards everything except `reports.code_catalog`. A
successful overwrite can therefore replace an SVG with a placeholder without the warning
required by the approved plan.

Carry the image report and diagram pass through successful completion of the local
overwrite. Emit `images:`, `svg:`, and applicable warning lines exactly once, with the
same semantics as ordinary create, including the renderer error when present. Cover both
filesystem Markdown overwrite and vault-note `-M` overwrite. Preserve existing
source/catalog ownership, transaction behavior, and failure output; do not emit success
summaries before the overwrite commits successfully. Keep normal snapshot SVG behavior
unchanged. Test vector success, sibling fallback, and placeholder diagnostics through
the CLI using the established fake adapter/render seams where possible; include
assertions for original source-directory resolution.

## 4. Verify the corrections

Run focused collector/prepare, adapter, and overwrite regressions, then `just check` and
`just check-web-clip-adapter`. Do **not** run `just check-full`. Use `/sase_monitor` if
a check escalates or needs a long-running continuation; handoff that existing run rather
than starting a duplicate. Record real tool skips. The lander has Pandoc and Chrome, but
no Poppler commands on PATH; do not claim gated PDF assertions executed if they skipped.
Run the existing foreignObject label/vector/no-raster acceptance wherever its tools are
already available, without installing host packages as part of this tale.

No documentation rewrite is required to establish these existing contract promises. If
documentation does need editing, read `docs.md` with `/sase_memory_read` first and run
`just docs-check`. The lander's current `just docs-check` fails only on the five known
mounted-page budgets below, which are independent of this epic.

## 5. Finish bob-cli-k6's landing in this turn

This is the tale's final step, not a later land-agent task. Before closing, re-read the
epic and its two children, verify both phase scopes and these repairs, and inspect
commits added since `7eb82440` for necessary SVG integration. The original epic has **no
parent_bead**, as verified during the landing review.

The lander already completed all PROPOSED FOLLOW-UP triage using `/sase_new_task`:

- **k6.1 note #1 and k6.2 note #1, four mounted pages:** corroborated existing
  **bob-cli-jz** with a fresh `just docs-check` at `7eb82440`: capture-panel 6036,
  capture-previews 6177, keyboard 6060, navigation-hotkeys 6186 words (cap 6000).
- **Both proposals, protocol.md:** corroborated existing **bob-cli-h9** at 6149 words.
  No new task; duplicate search, recent-task sweep, and active-epic scope review found
  no new causal owner for these mounted-source failures.
- **k6.2 note #1, json-outputs.md:** declined as already resolved. `ba3f51ba` explicitly
  trims this page, and the fresh docs check has no json-outputs error.

Include those outcomes in the epic close note; do not recreate or re-triage them unless
new evidence changes the result. The lander recorded the same audit and outcomes on
bob-cli-k6 before proposing this tale.

Run `sase bead epic-symbols bob-cli-k6`. It was empty at review, but recheck it. Resolve
every listed entry by wiring, privatizing, adding a justified non-test pragma, or
deleting the symbol; only re-key an exemption to an actually open later bead that still
needs it. Then close normally:

```sh
sase bead close bob-cli-k6 --note "Reviewed both phase scopes and every note, epic decisions and commits, intervening changes, and landing repairs; verified SVG collection, root sanitization, overwrite reporting and the relevant checks. Follow-up outcomes: k6.1 #1 and k6.2 #1 mounted budgets corroborated on jz/h9; k6.2 json-outputs proposal declined as fixed by ba3f51ba. Recapture=yes acceptance recorded by k6.2. No remaining epic-symbol exemptions."
```

Replace or extend that note with actual test results and any newer integration. If close
rejects leftover symbols, finish their cleanup and retry. If it rejects unfinished
phases, finish/reopen them and check descendant readiness; never use `--force` just to
succeed or to advance a successful nested landing.

After closing, run `just symvision` if the recipe is available (there is no such recipe
at `7eb82440`; record that when still true). Open the plans sidecar with `/sase_repo`
(`sase repo open plans -r "Mark the completed SVG epic plan done"`), read its
instructions, and set **`status: done`** in the frontmatter of
`202611/ref_create_svg_images.md`. Use the opened path, never a numbered workspace path
from another turn. The original epic has no parent to close. Finish through
`/sase_final`, including commit decisions for every modified repository. This closeout
must not depend on a post-turn commit, push, or CI result.
